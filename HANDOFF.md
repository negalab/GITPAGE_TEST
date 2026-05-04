# HANDOFF — Build a personal wiki + dual GitHub Pages publishing system

> **For an AI agent in a fresh session.** This document is a portable build guide. Follow it top-to-bottom to produce a working system in ~2 hours.
>
> **Reference implementation**: [github.com/negalab/GITPAGE_TEST](https://github.com/negalab/GITPAGE_TEST) — clone it to copy any file verbatim. All "see reference" pointers below mean "look at that repo."

---

## 0. What you're building

A **personal wiki + dual GitHub Pages publishing system**:

```
Single source of truth:  /WIKI/*.md  (Obsidian vault)
                              │
                              ▼
            scripts/sync-wiki.mjs  (frontmatter-aware fan-out)
                  │                          │
                  ▼                          ▼
   sites/public/  (Astro Fuwari)    sites/personal/  (Quartz v4)
   ── publish: true only            ── all WIKI files
   ── Helmut Newton editorial       ── obsidian wiki-link/graph
      design (cream paper, dark
      ink, Cormorant + Pinyon
      Script + Korean serif fb)
                  │                          │
                  └──────── single ──────────┘
                  GitHub Actions workflow
                  combines into one Pages site:
                  / (public) and /notes/ (personal)
```

Live URLs after deploy:
- **Public**: `https://${GITHUB_USERNAME}.github.io/${REPO_NAME}/`
- **Personal**: `https://${GITHUB_USERNAME}.github.io/${REPO_NAME}/notes/`

---

## 1. Variables to collect from the user — ASK FIRST

Before doing anything, run this short intake. Defaults in parentheses are sensible if the user says "알아서" / "your call":

| # | Variable | Question | Default |
|---|----------|----------|---------|
| 1 | `${BRAND}` | Site title / personal brand name? | (their handle / GitHub username) |
| 2 | `${TAGLINE}` | One-sentence tagline (will appear under the masthead)? | "personal notes & writing" |
| 3 | `${BIO}` | 1–2 sentence bio (will appear in profile sidebar)? | "writer · reader · maker" |
| 4 | `${GITHUB_USERNAME}` | Your GitHub username? | — required |
| 5 | `${GIT_EMAIL}` | Email for git commits? | — required |
| 6 | `${REPO_NAME}` | Repository name? | "wiki" or `${BRAND}-wiki` |
| 7 | `${SITE_LANG}` | Primary language (`en`, `ko`, `ja`, etc.)? | "en" |
| 8 | `${KOREAN_FALLBACK}` | If site has CJK text — use Noto Serif KR fallback? | yes if `${SITE_LANG}` is `ko`, else no |
| 9 | `${PROJECT_PATH}` | Where on disk should I create this? | `~/${REPO_NAME}` |

**Don't ask anything else upfront.** All other decisions are documented in §3 with sensible defaults; only confirm those if the user wants to override.

---

## 2. Prerequisites — verify before starting

```bash
node --version       # need 20+
npm --version        # need 9+
git --version        # any
gh --version         # 2.50+ recommended
```

**`gh` CLI auth** — must be logged into `${GITHUB_USERNAME}`. If user has multiple gh accounts:

```bash
gh auth status
```

If `${GITHUB_USERNAME}` not present:
```bash
gh auth login         # interactive — user does this in browser
```

If present but not active:
```bash
gh auth switch --user ${GITHUB_USERNAME}
```

> **Common pitfall**: `gh auth login --user ${GITHUB_USERNAME}` is invalid syntax. The `--user` flag exists only on `gh auth switch`, not `gh auth login`. Just run `gh auth login` (no flag) and the prompts handle it.

---

## 3. Default decisions (don't ask unless user wants to override)

| # | Decision | Default | Why |
|---|----------|---------|-----|
| 1 | All files | `.md` | Obsidian native |
| 2 | RAW originals after move | Move to `RAW/_archived/`, never delete | Preserves recoverability |
| 3 | Public-page selection | frontmatter `publish: true` flag (default `false`) | Simple, opt-in |
| 4 | Both sites public | yes (free GitHub Pages) | Private requires GitHub Pro |
| 5 | URL structure | Single repo, public at `/`, personal at `/notes/` | One workflow, simpler |
| 6 | SSG public | Astro **Fuwari** | Customizable + Astro ecosystem |
| 7 | SSG personal | **Quartz v4** | Native Obsidian wiki-link/graph |
| 8 | Personal site fonts | **Pretendard** (CJK) or **Inter** (Latin) | Modern, multi-weight |
| 9 | Public site design | **Helmut Newton editorial** (cream paper, Cormorant SC, Pinyon Script) | Magazine aesthetic from claude.ai/design |
| 10 | Newsletter | **Buttondown** placeholder embed | Free 100 subs, easy to swap |
| 11 | Korean text fallback | **Noto Serif KR** (paired with Cormorant for editorial) | Editorial tone for CJK |
| 12 | Categorization | Auto-classify by content (rules in CLAUDE.md) | No manual tagging burden |
| 13 | INDEX layout | Latest-first table + per-category sections | Two browse modes |

---

## 4. Build phases — execute in order

Each phase ends with a verification step. Don't move on until verified.

### Phase 1 — Foundation files (10 min)

```bash
cd ${PROJECT_PATH}
mkdir -p RAW/_archived WIKI sites scripts .github/workflows
touch RAW/.gitkeep RAW/_archived/.gitkeep WIKI/.gitkeep
```

Write 4 governance docs at the project root:
- **`CLAUDE.md`** — agent rules: trigger words ("정리해", "공개해", or English equivalents like "organize", "publish"), category rules, frontmatter spec with `publish: false` default, INDEX update protocol, what NOT to do
- **`README.md`** — user-facing usage docs
- **`INDEX.md`** — initial empty index with table headers + category sections (🔓 = published, 🔒 = personal-only)
- **`.gitignore`** — see Phase 4

Write 1–2 sample WIKI files so the build pipeline has something to render:
- `WIKI/welcome.md` with `publish: true` (intro for visitors)
- `WIKI/${REPO_NAME}-system.md` with `publish: false` (system meta, personal-only)

Frontmatter schema for ALL `WIKI/*.md` files:
```yaml
---
title: "..."
category: "AI|Tech|Business|Education|Content|Personal|Reference|Idea"
tags: [t1, t2, t3]
raw_source: "RAW/_archived/..."   # empty string if not from RAW
created: "YYYY-MM-DD"
updated: "YYYY-MM-DD"
summary: "..."
publish: false                     # CRITICAL default. User must opt in per file.
cover: ""
---
```

Reference: `CLAUDE.md`, `README.md`, `INDEX.md`, `WIKI/welcome.md` in the reference repo.

**Verify**: `ls -la` shows the 3 dirs + 4 governance docs + sample WIKI files.

---

### Phase 2 — Clone the two SSGs (5 min)

```bash
cd sites
git clone --depth 1 https://github.com/saicaca/fuwari.git public
git clone --depth 1 https://github.com/jackyzha0/quartz.git personal
rm -rf public/.git public/.github personal/.git personal/.github
cd ..
```

**Customize Fuwari** (`sites/public/src/config.ts`):
- `siteConfig.title`: `"${BRAND}"`
- `siteConfig.subtitle`: `"${TAGLINE}"`
- `siteConfig.lang`: `"${SITE_LANG}"`
- `siteConfig.themeColor`: `{ hue: 30, fixed: true }` (becomes irrelevant after editorial CSS hijacks `--primary`)
- `siteConfig.banner.enable`: `false` (the editorial masthead replaces it)
- `profileConfig.name`: `"${BRAND}"`
- `profileConfig.bio`: `"${BIO}"`
- `profileConfig.avatar`: `"assets/images/${BRAND_SLUG}-monogram.svg"` (created in Phase 6)
- `profileConfig.links`: GitHub link to `https://github.com/${GITHUB_USERNAME}`
- `navBarConfig.links`: Home, Archive, About + external `{ name: "📓 Notes", url: "/${REPO_NAME}/notes/", external: true }`

**Customize Fuwari** (`sites/public/astro.config.mjs`):
- `site`: `"https://${GITHUB_USERNAME}.github.io"`
- `base`: `"/${REPO_NAME}/"`

**Clear Fuwari demo posts**:
```bash
rm -rf sites/public/src/content/posts/*
touch sites/public/src/content/posts/.gitkeep
```

**Customize Quartz** (`sites/personal/quartz.config.ts`):
- `pageTitle`: `"${BRAND} Notes"`
- `analytics`: `null`
- `locale`: `"${SITE_LANG}-${COUNTRY}"` (e.g. `"ko-KR"`, `"en-US"`)
- `baseUrl`: `"${GITHUB_USERNAME}.github.io/${REPO_NAME}/notes"`
- `theme.typography`: `{ header: "Inter", body: "Inter", code: "JetBrains Mono" }`
  > **DO NOT use "Gowun Dodum"** or any single-weight Google Font — Quartz tries to fetch weight 700 and the build fails. Use Inter (or any font with all weights 400/500/600/700 on Google Fonts).

**Customize Quartz** (`sites/personal/quartz/styles/custom.scss`):
```scss
@use "./base.scss";

// Web font is loaded via <link> in Head.tsx — NOT @import here.
// @import url(...) BREAKS the build because it must precede all rules,
// but base.scss above already emits CSS rules.

:root {
  --bodyFont: "Pretendard Variable", Pretendard, -apple-system, BlinkMacSystemFont,
              system-ui, "Apple SD Gothic Neo", "Noto Sans KR", sans-serif;
  --headerFont: "Pretendard Variable", Pretendard, -apple-system, system-ui, sans-serif;
}
body { font-family: var(--bodyFont); }
h1, h2, h3, h4, h5, h6 { font-family: var(--headerFont); font-weight: 700; letter-spacing: -0.02em; }
article p, article li { line-height: 1.75; }
```
(For non-CJK sites, swap Pretendard for Inter.)

**Inject Pretendard into Quartz Head** (`sites/personal/quartz/components/Head.tsx`):
Find the `<link rel="preconnect" href="https://cdnjs.cloudflare.com" ... />` line and insert AFTER it:
```jsx
<link
  rel="stylesheet"
  href="https://cdn.jsdelivr.net/gh/orioncactus/pretendard@v1.3.9/dist/web/variable/pretendardvariable-dynamic-subset.min.css"
/>
```

**Verify**: configs edited and saved.

---

### Phase 3 — Sync script (10 min)

Write `scripts/sync-wiki.mjs`. Behavior:
1. Read all `WIKI/*.md`
2. For each, parse frontmatter
3. Write to `sites/personal/content/{file}.md` always (Quartz reads our frontmatter natively)
4. If `publish === true`, also write to `sites/public/src/content/posts/{file}.md` with frontmatter rewritten to Fuwari's schema
5. Generate `sites/personal/content/index.md` listing all entries (Quartz needs a homepage)

**Critical detail #1 — date emission**:
Fuwari's content collection schema parses `published` as **Date type**. Quoted strings fail validation:
```yaml
published: 2026-05-04        # ✅ correct
published: "2026-05-04"      # ❌ Astro fails: "Expected type 'date', received 'string'"
```
In your serializer, detect `/^\d{4}-\d{2}-\d{2}$/` and emit bare (no quotes).

**Critical detail #2 — Fuwari frontmatter mapping**:
| WIKI field | Fuwari field |
|-----------|--------------|
| `title` | `title` |
| `created` | `published` (bare date) |
| `summary` | `description` |
| `tags` | `tags` |
| `category` | `category` |
| `publish: true` | `draft: false` |
| `cover` (if non-empty) | `image` |

**Critical detail #3 — `content/index.md` generation**:
Quartz fails to build without `content/index.md`. The sync clears the dir each run, so a static index.md gets wiped. Your sync script **must write index.md after copying** all entries — list them with publish-status badges (🔓 / 🔒) using Obsidian wiki-link `[[slug|title]]` syntax.

Reference: `scripts/sync-wiki.mjs` in the reference repo. ~130 LOC, no external deps (regex-based YAML parser).

**Verify**:
```bash
node scripts/sync-wiki.mjs
# expect:
# ✓ Synced N files → sites/personal/content/
# ✓ Generated content/index.md with N entries
# ✓ Synced M files → sites/public/src/content/posts/ (publish: true only)
```

---

### Phase 4 — GitHub Actions workflow + .gitignore (5 min)

Write `.github/workflows/deploy.yml`. Single job that:
1. Checkout
2. Setup Node 22
3. Setup pnpm 9 (Fuwari uses pnpm; lockfile is `pnpm-lock.yaml`)
4. Run `node scripts/sync-wiki.mjs`
5. `cd sites/public && pnpm install --frozen-lockfile && pnpm build`
6. `cd sites/personal && npm ci && npx quartz build` (Quartz uses npm; lockfile `package-lock.json`)
7. Combine: `cp -r sites/public/dist/* _site/` then `mkdir _site/notes && cp -r sites/personal/public/* _site/notes/`
8. `actions/upload-pages-artifact@v3` → `actions/deploy-pages@v4`

Permissions: `contents: read`, `pages: write`, `id-token: write`. Concurrency group: `pages`.

Reference: `.github/workflows/deploy.yml` in the reference repo (~65 LOC).

`.gitignore`:
```
.DS_Store
node_modules/
sites/public/dist/
sites/public/.astro/
sites/personal/public/
sites/personal/.quartz-cache/
*.log
.env*
```

---

### Phase 5 — Init git, create repo, push 🚨 USER CONFIRMATION REQUIRED

```bash
cd ${PROJECT_PATH}
git init -b main
git config user.email "${GIT_EMAIL}"
git config user.name "${GITHUB_USERNAME}"
git add -A
git commit -m "Initial commit — RAW→WIKI pipeline + two-site Pages publishing"
```

🚨 **STOP HERE — confirm with user before pushing.** This makes the work public on the internet. State explicitly what will be public (the welcome.md content, the system files, their bio) and ask: "이대로 인터넷에 공개됩니다. push할까요?" / "This will be live on the internet. OK to push?"

After explicit yes:
```bash
gh repo create ${GITHUB_USERNAME}/${REPO_NAME} --public --source=. --remote=origin --push --description "Personal wiki + two-site GitHub Pages publishing"
gh api -X POST /repos/${GITHUB_USERNAME}/${REPO_NAME}/pages -f 'build_type=workflow'
```

The `gh api` call enables Pages with GitHub Actions as source.

**Watch the first build** — first builds may fail because of issues in §6:
```bash
until [ "$(gh run list --repo ${GITHUB_USERNAME}/${REPO_NAME} --limit 1 --json status --jq '.[0].status')" = "completed" ]; do sleep 15; done
gh run view --repo ${GITHUB_USERNAME}/${REPO_NAME} --log-failed
```

---

### Phase 6 — Apply Helmut Newton editorial design

The default Fuwari design is a generic dark-mode-friendly blog template. Replace the visual layer with the editorial design (cream paper + dark ink + Cormorant SC + Pinyon Script).

**Strategy**: Don't restructure Fuwari's components. Layer a CSS override on top + add a magazine masthead component for the home page only. Much safer and faster than rewriting Fuwari.

#### 6a. Add fonts to Fuwari head

Edit `sites/public/src/layouts/Layout.astro`. Inside `<head>`, after `<meta name="generator">`:

```html
<link rel="preconnect" href="https://fonts.googleapis.com" />
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin />
<link href="https://fonts.googleapis.com/css2?family=Cormorant+Garamond:ital,wght@0,400;0,500;0,600;0,700;1,400;1,500;1,600&family=Cormorant+SC:wght@500;600;700&family=Libre+Caslon+Text:ital,wght@0,400;0,700;1,400&family=Pinyon+Script&family=Noto+Serif+KR:wght@400;500;600;700&display=swap" rel="stylesheet" />
```
(Drop `Noto+Serif+KR` if `${SITE_LANG}` is not `ko`.)

#### 6b. Editorial design system in `sites/public/src/styles/main.css` (top)

Replace any existing font/`:root` block with:

```css
:root {
    --paper: #f6f3ec;
    --paper-deep: #d6d2c8;
    --ink: #161410;
    --ink-soft: #1d1a13;
    --rule: #1a1a1a;
    --muted: #5a5750;
    --headline-tracking: -0.01em;
    --body-tracking: 0.005em;
    --label-tracking: 0.32em;
    --body-leading: 1.65;
    --headline-leading: 0.95;

    --font-serif: "Cormorant Garamond", "Noto Serif KR", "Libre Caslon Text", Georgia, serif;
    --font-serif-sc: "Cormorant SC", "Cormorant Garamond", "Noto Serif KR", serif;
    --font-script: "Pinyon Script", "Italianno", cursive;
    --font-label: "Libre Caslon Text", "Cormorant Garamond", "Noto Serif KR", serif;
}

html, body {
    font-family: var(--font-serif);
    background: var(--paper-deep);
    color: var(--ink);
    -webkit-font-smoothing: antialiased;
}
```

#### 6c. Append editorial overrides to bottom of `main.css`

A ~270 LOC override section that:
- Forces `html, body` background to `--paper-deep`
- Flattens `.card-base` (no rounded corners, no shadow, paper bg, top border)
- Restyles post card titles → Cormorant SC uppercase
- Hijacks `--primary` → `var(--ink)` so all Fuwari accent colors map to ink
- Restyles `#navbar` → top/bottom 1px rules, label font, uppercase
- Applies `filter: grayscale(1) contrast(1.05)` to all post images
- Restyles `.custom-md` (article body): first paragraph as lede (centered italic), h1 as huge SC, h2 as section label with bottom rule, blockquote as pull quote, hr as 33%-width centered rule
- Drops rounded corners from buttons/panels/pagination
- Footer in label font

Reference: `sites/public/src/styles/main.css` lines 187+ in the reference repo. Copy verbatim.

#### 6d. Editorial masthead component

Create `sites/public/src/components/EditorialMasthead.astro`. Three-column grid (badge-left | title-stack | badge-right). Title stack: `${BRAND}` first half (huge SC) / Pinyon Script accent / `${BRAND}` second half (huge SC). Side badges in italic. Below: lede with `${BIO}`. Bottom: thin rule. Mobile: collapse to single column.

Reference: `sites/public/src/components/EditorialMasthead.astro` (~145 LOC).

#### 6e. Inject masthead on home page only

Edit `sites/public/src/pages/[...page].astro`:
```jsx
import EditorialMasthead from "../components/EditorialMasthead.astro";
// ...
<MainGridLayout>
  {page.url.prev === undefined && <EditorialMasthead />}
  <div class="editorial-section-label-wrap"><span class="editorial-label">LATEST DISPATCHES</span></div>
  <PostPage page={page}></PostPage>
  <Pagination ... />
  {page.url.prev === undefined && <Newsletter />}
</MainGridLayout>
```
The `page.url.prev === undefined` check ensures the masthead shows only on page 1, not paginated archive pages.

#### 6f. Editorial Newsletter component

Rewrite `sites/public/src/components/Newsletter.astro` with magazine styling: 3-column grid (badge | body | badge), thick top rule, Cormorant SC headline + Pinyon Script accent, lede paragraph, ink-on-paper button. Buttondown form with placeholder username `YOUR_USERNAME_HERE` — recipient activates by signing up at buttondown.com and replacing the constant.

Reference: `sites/public/src/components/Newsletter.astro` (~175 LOC).

---

### Phase 7 — Replace placeholder avatar (5 min)

Fuwari's `demo-avatar.png` is an anime cartoon — clashes with editorial design. Replace with a self-contained SVG monogram.

Write `sites/public/src/assets/images/${BRAND_SLUG}-monogram.svg`:
- 400×400 viewBox
- Cream paper background + dot pattern + subtle grain filter
- Top + bottom horizontal rules
- Top label: `"VOL. I · ISSUE 001"` in italic serif
- Center: `${BRAND}` initials in serif (`Georgia` family)
- Below: `"${BRAND} subtitle"` or any tagline in cursive (`'Brush Script MT', 'Snell Roundhand', cursive`)
- Bottom: `${TAGLINE}` and date

🚨 **Critical**: SVGs loaded via `<img src="*.svg">` render in their own context with NO access to web fonts loaded by parent HTML. Use only generic font families (`serif`, `cursive`) or system fonts (`Georgia`, `'Brush Script MT'`). Cormorant SC / Pinyon Script will NOT render here.

Update `sites/public/src/config.ts` so `profileConfig.avatar` points to the new SVG.

Append to `main.css`:
```css
#sidebar .card-base a[href*="/about/"] {
    border-radius: 0 !important;
    border: 1px solid var(--rule);
}
#sidebar .card-base a[href*="/about/"] img {
    filter: none !important;     /* SVG is already monochrome */
    border-radius: 0 !important;
}
#sidebar .card-base .font-bold {
    font-family: var(--font-serif-sc) !important;
    text-transform: uppercase;
}
#sidebar .card-base .h-1 { display: none !important; }
```

Reference: `sites/public/src/assets/images/mr5pm-monogram.svg` (~50 LOC).

---

## 5. Final commit + push

```bash
node scripts/sync-wiki.mjs
git add -A
git commit -m "Apply editorial design"
git push
```

Watch the build. If it fails, see §6.

---

## 6. Build failures to expect (and the fixes)

Even with this guide, the first 1–3 builds often fail. **Avoid by writing correct code from the start.** If they recur:

| # | Symptom | Root cause | Fix |
|---|---------|-----------|-----|
| 1 | Astro: `data does not match collection schema. published: Expected type "date", received "string"` | Sync emitted `published: "2026-05-04"` (quoted) | In serializeFrontmatter, detect `/^\d{4}-\d{2}-\d{2}$/` and emit bare |
| 2 | Quartz: `Failed to fetch font X with weight 700, got Bad Request` | Google Fonts has X at fewer weights | Switch typography to Inter or another font with weights 400/500/600/700 |
| 3 | Quartz: `Failed to emit from plugin ComponentResources: @import rules must precede all rules` | Put `@import url(...)` AFTER `@use "./base.scss"` in custom.scss | Move web font loading to `<link>` in Head.tsx; SCSS can't have `@import url()` after other rules |
| 4 | Quartz: `Warning: missing index.md home page file` → build still fails downstream | Sync wiped content/ each run, no homepage | Have sync script generate content/index.md after copying entries |
| 5 | Astro/Pages: 404 on root URL after deploy | `base` in astro.config.mjs doesn't match repo path | Set `base: "/${REPO_NAME}/"` exactly |

---

## 7. Verification checklist

After final push and successful build:

```bash
curl -sI https://${GITHUB_USERNAME}.github.io/${REPO_NAME}/      | head -1   # 200
curl -sI https://${GITHUB_USERNAME}.github.io/${REPO_NAME}/notes/ | head -1   # 200

# editorial markers present
curl -s https://${GITHUB_USERNAME}.github.io/${REPO_NAME}/ | grep -oE '(Cormorant|Pinyon|'"${BRAND}"')' | sort -u
```

Visual checks (open both URLs):
- ✅ Public: cream background, brand masthead at top, "LATEST DISPATCHES" label, post cards as flat magazine entries with serif uppercase titles, Newsletter card at bottom
- ✅ Public: profile avatar shows the SVG monogram (NOT the anime cartoon)
- ✅ Personal: Quartz default layout, body font is the chosen Latin/CJK font, content index page with 🔓/🔒 badges
- ✅ Both: post images grayscale on public, normal on personal

---

## 8. General principles to follow (good defaults)

- ❌ Don't auto-push without explicit user confirmation (public action)
- ❌ Don't auto-flip `publish: false` → `true` without user saying "publish [filename]"
- ❌ Don't delete RAW originals — always move to `_archived/`
- ❌ Don't ship a single-default for visual choices that the user will SEE — present 3–5 options with previews
- ✅ DO ship working pipeline first, polish later — ugly-but-working > pretty-but-broken
- ✅ DO reference existing files in the reference repo verbatim where possible — don't reinvent

---

## 9. Things to ASK the user (only these)

1. **Phase 0**: the §1 intake (9 variables)
2. **Phase 5**: explicit confirmation before `gh repo create` + first push (public action)
3. **If gh CLI auth is missing or wrong account active**: walk them through `gh auth login`
4. **If a WIKI file collides with same target name during sync**: ask "merge / new file / skip?"
5. **If a category is unclear during RAW→WIKI organize**: present 2 candidates

Everything else: assume from §3 defaults. Do not ask which SSG, which fonts, which design — those are decided.

---

## 10. File reference (all paths relative to project root)

| Path | Purpose | LOC ≈ |
|------|---------|-------|
| `CLAUDE.md` | Agent rules: triggers, frontmatter spec, INDEX update protocol | 200 |
| `README.md` | User-facing usage docs | 130 |
| `INDEX.md` | Two-section content index, manually + sync-updated | 50 |
| `HANDOFF.md` | This file | — |
| `RAW/` | User inbox (.gitkeep) | — |
| `RAW/_archived/` | Originals after sync (.gitkeep) | — |
| `WIKI/welcome.md` | Sample post, publish: true | 30 |
| `scripts/sync-wiki.mjs` | WIKI → both sites with publish flag filter | 130 |
| `sites/public/` | Astro Fuwari (cloned + customized) | — |
| `sites/public/src/config.ts` | Site title, profile, navbar | 80 |
| `sites/public/astro.config.mjs` | site URL + base path | 170 |
| `sites/public/src/styles/main.css` | Base + editorial overrides | 480 |
| `sites/public/src/layouts/Layout.astro` | Adds editorial fonts to head | 570 |
| `sites/public/src/components/EditorialMasthead.astro` | Magazine masthead (home only) | 145 |
| `sites/public/src/components/Newsletter.astro` | Editorial Buttondown card | 175 |
| `sites/public/src/pages/[...page].astro` | Home page; injects masthead + label + newsletter | 30 |
| `sites/public/src/assets/images/*-monogram.svg` | SVG avatar | 50 |
| `sites/personal/` | Quartz v4 (cloned + customized) | — |
| `sites/personal/quartz.config.ts` | Site title, baseUrl, fonts (Inter), locale | 100 |
| `sites/personal/quartz/styles/custom.scss` | Pretendard typography overrides | 25 |
| `sites/personal/quartz/components/Head.tsx` | Pretendard `<link>` injection | 110 |
| `.github/workflows/deploy.yml` | Sync + build both + combine + deploy Pages | 65 |
| `.gitignore` | Excludes node_modules, builds, .DS_Store, .env | 30 |

---

## 11. User's day-to-day rhythm (after build is done)

1. **Add content**: drop `.md` file into `RAW/`
2. **Trigger organize**: tell Claude "organize" / "정리해" — Claude reads `CLAUDE.md` rules, parses RAW files, writes structured `WIKI/{slug}.md` with `publish: false`, moves originals to `RAW/_archived/`, updates INDEX.md
3. **Selectively publish**: tell Claude "publish {slug}" / "공개해 {slug}" or directly edit `publish: true` — only flagged files appear on the public site
4. **Deploy**: `node scripts/sync-wiki.mjs && git add -A && git commit -m "..." && git push` — GitHub Actions builds both sites in 5–7 min

---

## 12. If you (the AI) get stuck

- Read the reference repo: [github.com/negalab/GITPAGE_TEST](https://github.com/negalab/GITPAGE_TEST). Every file referenced in this doc exists there with working contents.
- Build failures: check §6 first — the same 5 issues recur across rebuilds.
- Visual choices: present multiple options with previews; never install one default and stop.
- Auth/push/destructive actions: always confirm with user before executing.
- When in doubt about a user preference not covered in §3: ask once, briefly, with 2–3 options.

---

**End of handoff.** A fresh AI following this document with a willing user should produce a functionally identical system in ~2 hours, with two user pauses: §1 intake (5 min) and §5 push confirmation (1 min).
